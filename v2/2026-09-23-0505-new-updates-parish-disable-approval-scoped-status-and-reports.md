# 2026-09-23 05:05 — Parish disable by approval, scoped status, and two reports

**Branch:** `dev` · **Verified:** 1193 passing

Four changes. The first one **takes away an ability people have today**, so read
that section even if you skip the rest.

---

## Behaviour changes on deploy

| Change | Effect |
|---|---|
| **An ordinary administrator can no longer disable a parish** | `DELETE /v1/parishDirectory/:id` returns `403 PARISH_DISABLE_NEEDS_APPROVAL` and hands them the request endpoint. Only super-admin and national support may still act directly |
| **`PATCH /v1/users/:id/status` is now scoped, not super-admin-only** | a province admin can deactivate a leaver in their own province. This is a *widening* |
| **An approved parish disable expires after six months** | it reactivates on its own unless somebody acts |

---

## 1. Disabling a parish is a request now

### Why

A parish is where people worship. Taking one out of service is a decision about
a congregation, not an administrative tidy-up — so an administrator **asks**,
and a super-admin **answers**.

And a closure is never open-ended. A parish disabled and forgotten is worse than
one closed deliberately: it simply disappears from the organisation with nobody
accountable for the decision having lapsed. Every approved disable therefore
carries a `reactivateAt` six months out, and a cron brings it back.

### Raising one

```http
POST /v1/approvals/parish-disable
Authorization: Bearer <token>
Content-Type: application/json

{
  "parishCode": "211003",
  "reason": "Congregation merged into the parish next door."
}
```

`reason` must be **at least ten characters**. Somebody will ask why this parish
closed, months later, and "nobody wrote it down" is not an answer.

**201**

```json
{
  "requestType": "PARISH_DISABLE",
  "status": "PENDING",
  "subjectUnitCode": "211003",
  "scopeCode": "LA47",
  "reason": "Congregation merged into the parish next door.",
  "requestedByUsername": "prov.admin",
  "planSnapshot": {
    "parishName": "RCCG EXAMPLE PARISH",
    "provinceCode": "LA47",
    "areaCode": "A31",
    "zoneCode": "Z12",
    "memberCount": 42
  }
}
```

`planSnapshot.memberCount` is recorded **now**, because it is what the approver
needs to weigh and it will have changed by the time anyone looks back.

**400** — no reason, reason too short, unknown parish, already disabled, or one
already pending for that parish.

### Who decides

**Super-admin only.** Not national support, and not the province that asked.

That check sits *above* the `isUnbounded()` shortcut in `canDecide`, which would
otherwise have admitted the elevated roles — there is a test for exactly that,
because it is the kind of thing a later refactor quietly undoes.

```http
POST /v1/approvals/{id}/approve
POST /v1/approvals/{id}/reject
```

### What approval does

- `deletedAt` set, `status` → `"0"`, so every `ACTIVE_STATUS` filter skips it
- `disabledReason` carried over from the request
- `reactivateAt` = now + 6 months
- **the row stays**, so the parish's members still resolve a chain and are not
  orphaned by the closure

### The refusal an administrator now sees

```json
{
  "code": "PARISH_DISABLE_NEEDS_APPROVAL",
  "message": "Disabling a parish is a request now, not an action — it closes a place people worship in, so a super-admin decides. Raise it with a reason and it reactivates automatically after six months unless somebody acts.",
  "useInstead": {
    "useEndpoint": "/v1/approvals/parish-disable",
    "body": { "parishCode": "211003", "reason": "" }
  }
}
```

### The reactivation cron

`src/config/cron/parishReactivation.ts`, daily at 03:40. Clears the deletion
marks, `status` back to `"1"`, and writes a `PARISH_REACTIVATED` activity line
naming the original reason. Idempotent — a parish already back no longer matches.

| Variable | Default | Notes |
|---|---|---|
| `PARISH_REACTIVATION_ENABLED` | *(off)* | **must be set to `true`** or disabled parishes never come back |
| `PARISH_REACTIVATION_CRON` | `40 3 * * *` | daily; a day's precision is plenty for a six-month timer |

> If you enable the disable flow but **not** this cron, every closure is
> permanent in practice. Turn both on together.

---

## 2. Setting a user's status is scoped

`PATCH /v1/users/:id/status` was super-admin only, so a province administrator
could not deactivate a leaver in their own province without raising a ticket.

It now judges the caller's standing with **`canDeleteUser`** — deliberately the
same bar as deletion. Setting a status is a *lesser* act than deleting, so
anyone permitted to delete an account is certainly permitted to deactivate it,
and one rule governs both rather than two drifting apart.

That brings the protections with it: super-admin and the elevated roles are not
touchable by a scoped actor, the target must sit inside the caller's own unit,
and you still cannot deactivate yourself.

**403** carries the reason and a code (`TARGET_PROTECTED`, `OUTSIDE_UNIT`, …);
self-deactivation is a `400`.

---

## 3. Who has been deleting

```http
GET /v1/deletions/actors?module=PARISH_DIRECTORY&from=2026-09-01&to=2026-09-30
```

The listing at `/v1/deletions` says *what* was deleted. This says *who is doing
it* — the question asked when a number looks wrong. One account removing four
hundred parishes in a week is a different conversation from forty administrators
tidying up ten each.

```json
{
  "count": 2,
  "actors": [
    {
      "deletedBy": "64f0aa...",
      "deletedByUsername": "tunde.support",
      "module": "PARISH_DIRECTORY",
      "deletions": 412,
      "firstAt": "2026-09-14T08:02:11.000Z",
      "lastAt": "2026-09-14T08:49:55.000Z",
      "samples": ["211003", "211004", "211005"]
    }
  ]
}
```

`firstAt`/`lastAt` are there so a burst reads as a burst — 412 deletions inside
47 minutes is a different story from 412 across a month. Scoped by the same
filter as the listing, so nobody sees deletions outside their own unit.

---

## 4. Sweep reports: export and resolutions

### `GET /v1/hierarchy-sync/runs/:id/export?format=csv`

CSV by default, because the orphan list gets worked through in a spreadsheet.
Every row carries its bucket **and the action for it**, so the file is still
useful away from this API.

```csv
bucket,parishCode,users,modified,detail,action
"STALE","211003","2","2","was {""province"":""LA30""} -> {""province"":""LA47""}","repaired by this run"
"ORPHAN","874112","1","0","parish code is in no directory row","Check whether the parish code on the users is a typo..."
"SPLIT","990001","1","0","2 variants disagree","Repair with POST /v1/hierarchy-transfers/admin/realign..."
```

`?format=json` returns the same rows with the run's metadata.

### `GET /v1/hierarchy-sync/resolutions`

What to do about each bucket: what it means, **why the job did not fix it
itself**, the steps that do fix it, and what not to do. Written once in the code
so the advice cannot drift from the behaviour.

Each bucket carries a `doNot`, because the tempting shortcuts are the damaging
ones:

- **ORPHAN** — *"Do not clear the codes to make the report look clean. A blank
  parish is not a fix."*
- **SPLIT** — *"Do not transfer the unit to force agreement — that moves it as
  well as repairing it."*
- **DIRECTORY_GAP** — *"Do not assume the users are wrong. Here, the directory is
  the incomplete side."*

---

## Tests updated, not just added

Two existing suites asserted the behaviour this change removes — that a plain
`DELETE` soft-deletes for any administrator. They now assert the 403 and the
approval hint, with a separate case for the elevated path that still acts
directly. `test/audit-attribution-db.js` needed a real elevated user rather than
an id in a token, because the guard reads the caller's live record.

---

## Known gaps

- **Nothing stops a super-admin approving a disable they raised themselves** for
  this type. The self-approval rule elsewhere in approvals should cover it;
  worth confirming before the first real use.
- **No endpoint lists parishes awaiting reactivation.** `GET /v1/parishDirectory/deleted`
  shows them with `reactivateAt`, which is enough for now but is not a scheduled
  view.
- **`GET /deleted` and `/restore` are still elevated-only, not scoped** to the
  caller's province.
