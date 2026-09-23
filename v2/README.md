# documentations/

Dated release notes for developers. One file per batch of changes, newest first.

Written for somebody who was not in the room: what changed, why, the endpoints
with real request and response bodies, and what they have to do about it.

## Naming

```
YYYY-MM-DD-HHMM-new-updates-<short-summary>.md
```

The date and time are when the work landed on `dev`. The summary is a few words
in kebab case. Example:

```
2026-09-23-0419-new-updates-user-and-parish-deletion-and-restore.md
```

## What goes in one

Each file carries, in this order:

1. **What changed and why** — the problem, not just the diff.
2. **Behaviour changes on deploy** — anything that acts differently without a
   flag being set. Put this near the top; it is what breaks people.
3. **Endpoints** — method, path, guard, request body, response body. Real
   payloads, captured from a run, not invented ones.
4. **Configuration** — new environment variables, defaults, and whether they
   need registering in `deployment-scripts/lib.sh`.
5. **Deployment steps** — index scripts, backfills, the order they run in.
6. **Known gaps** — what was deliberately left, and what is still unsafe.

## The index

| Date | File | Covers |
|---|---|---|
| 2026-09-23 | [user-and-parish-deletion-and-restore](2026-09-23-0419-new-updates-user-and-parish-deletion-and-restore.md) | Soft delete, restore and permanent delete for users and parishes; nat-support may rename |
| 2026-09-22 | [privileged-audit-and-hierarchy-sync](2026-09-22-1455-new-updates-privileged-audit-and-hierarchy-sync.md) | The privileged audit trail, the user-hierarchy sweep, two new principal offices |
| 2026-09-20 | [impersonation-sessions-and-deletion-records](2026-09-20-0414-new-updates-impersonation-sessions-and-deletion-records.md) | Borrowed sessions survive refresh, sensitive roles stripped not refused, the deletion record becomes readable |
| 2026-09-19 | [activity-log-archive-and-search](2026-09-19-0849-new-updates-activity-log-archive-and-search.md) | Archived activity history, the approvals inbox, indexed search, parish country codes |

## Standing references kept here

| File | Covers |
|---|---|
| [reference-transfer-jobs-approvals-and-rollback](reference-transfer-jobs-approvals-and-rollback.md) | How a hierarchy change is recorded, approved and undone; batching many moves; what cannot be undone |

## The longer references

These notes are a changelog. The living references stay where they are:

- [PRIVILEGED_AUDIT_DOCS.md](../PRIVILEGED_AUDIT_DOCS.md)
- [HIERARCHY_SYNC_DOCS.md](../HIERARCHY_SYNC_DOCS.md)
- [HIERARCHY_MOVES_GUIDE.md](../HIERARCHY_MOVES_GUIDE.md)
- [PRINCIPAL_OFFICERS_DOCS.md](../PRINCIPAL_OFFICERS_DOCS.md)
- [CONFIGURATION.md](../CONFIGURATION.md) — every environment variable
