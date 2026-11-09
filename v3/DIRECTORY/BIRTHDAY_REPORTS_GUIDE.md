# Birthday reports — operator guide

How to switch on the daily birthday reports, test them safely, and then move
from the test recipient to the real hierarchy recipients. The full reference
is in [BIRTHDAY_REPORTS_DOCS.md](./BIRTHDAY_REPORTS_DOCS.md).

## 1. Prerequisites

- **The indexes exist.** Run `scripts/createMessagingIndexes.ts`. It creates
  `uniq_birthday_report_delivery`, the index that prevents a rerun from
  emailing the same people twice. See the
  [messaging integration guide](./MESSAGING_INTEGRATION_GUIDE.md#0-before-anything-create-the-indexes).
- **Email works.** `POST /v1/messaging/email/send-test` returns `ACCEPTED`.
- **The cron task is the one with `CRON_ENABLED` on.** Production API tasks
  run with `CRON_ENABLED=false`, so the report must be enabled on the cron
  task.

## 2. Test mode first

Set these on the cron task:

```
BIRTHDAY_REPORTS_ENABLED=true
BIRTHDAY_REPORTS_TEST_RECIPIENT=thepremiumcoder@gmail.com
# optional — these are the defaults
BIRTHDAY_REPORTS_CRON=0 6 * * *
BIRTHDAY_REPORTS_TIMEZONE=Africa/Lagos
```

At boot the log says:
`[birthdayReports cron] Scheduled '0 6 * * *' (Africa/Lagos) — TEST mode, sending only to the test recipient.`

You don't have to wait for 06:00. An elevated user can run it straight away:

```
POST /v1/birthday/reports/run
{ "date": "2026-11-08" }
```

The test address receives one email per report type, ADMIN and MEMBER, for
each type that has celebrants that day. **Check the appendix** in each email:
it lists every officer, pastor in charge and super-admin who would get a
slice in live mode, and who is in that slice. That appendix is how you check
the recipient rules against real data before anyone else is emailed.

Running again the same day sends nothing new, because the deliveries are
already `SENT`. To send the same day again for another look, a super-admin
passes `"force": true`.

## 3. What to check before going live

- Every slice in the appendix matches the officer's own unit. An area admin
  should see only their own area's celebrants.
- Parishes with members have a recipient, either the pastor in charge or the
  directory's pastor email. A parish with neither gets no member report. Fix
  the data, or accept that.
- The super-admin list is the one you expect. Every super-admin gets both
  full reports every day.
- `GET /v1/birthday/reports?reportDate=…` shows the TEST deliveries as `SENT`.

## 4. Cut-over to hierarchy recipients

1. Remove `BIRTHDAY_REPORTS_TEST_RECIPIENT` from the cron task, so it is
   unset, not empty.
2. Deploy. The boot log now ends with `— LIVE mode.`
3. The next scheduled run emails the hierarchy.

   Test and live deliveries are keyed differently: test rows have `scopeCode`
   `TEST`, live rows have the real scope. So if you go live part-way through
   a day, a manual run the same day still reaches every live recipient once.
4. Watch `GET /v1/birthday/reports?reportDate=<today>&status=FAILED` for the
   first few days.

**To roll back**, set `BIRTHDAY_REPORTS_TEST_RECIPIENT` again, or set
`BIRTHDAY_REPORTS_ENABLED=false`.

## 5. Day-to-day

- **No celebrants, no email.** A silent day is normal.
- **A failed send** shows as `FAILED` with the provider's `error`. Running the
  reports again for that date retries only the failed and unsent ones.
- **A suppressed recipient** (when `EMAIL_SUPPRESSION_ENABLED` is on) is
  recorded as `SKIPPED` with reason `SUPPRESSED`. Fix the address on their
  profile, and release the suppression only if the old address really works.

## 6. Frontend impact of the new guards

Members (`rpms-member`) no longer have access to `/v1/birthday/templates` or
`/v1/birthday/logs` (they now get 403). This is intended. Administrative users
now see only their own parishes. The screens to change are listed in
[FRONTEND_INTEGRATION_NOTES.md](./FRONTEND_INTEGRATION_NOTES.md#birthdays).
