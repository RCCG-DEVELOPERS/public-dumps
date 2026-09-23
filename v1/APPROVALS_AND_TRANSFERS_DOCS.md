# Approvals & Transfers — API Reference

Officer rosters, level-scoped appointment, and the approval workflow for
transfers and promotions.

Companion to `PRINCIPAL_OFFICERS_DOCS.md` (the office register itself) and
`ROLES_API_DOCS.md` (active roles and secondary grants).

- [Who can do what](#who-can-do-what)
- [Seeing your officers](#seeing-your-officers)
- [Appointing at your own level](#appointing-at-your-own-level)
- [The approval workflow](#the-approval-workflow)
  - [Officer transfer](#post-v1approvalsofficer-transfer)
  - [Officer promotion](#post-v1approvalsofficer-promotion)
  - [User parish transfer](#post-v1approvalsuser-transfer)
  - [Deciding](#deciding)
  - [Listing](#listing)
- [Removing a secondary role](#removing-a-secondary-role)
- [Error codes](#error-codes)
- [Frontend guidance](#frontend-guidance)

---

## Who can do what

Authority is **standing**, not a permission list: you act at the levels your own
roles sit at, and only at your own unit within each. Someone holding
`prov-admin` acts on *their* province — not every province, and not at region
level.

`super-admin` is unbounded. So is **national support** (`nat-support`, via
`ELEVATED_ROLES`) — over the hierarchy, principal officers, HQ assignment and
approvals, and over nothing else. Wherever this document says "super-admin" as
an approver or actor, read "super-admin or national support".

| Action | Who |
|---|---|
| See the officers of a unit | Anyone with standing at that unit; super-admin / national support anywhere |
| Appoint to a **vacant** office | Anyone with standing at that unit; super-admin / national support anywhere |
| End or transfer a **sitting** officer | Super-admin / national support — or a request approved by one |
| Approve an officer transfer / promotion | **Super-admin / national support only** |
| Approve a user parish transfer | **Province admin** of either province involved, or super-admin / national support |
| **Raise** a unit (parish/area/zone) transfer into another province | **Province admin of the province the unit is LEAVING** |
| **Approve** a unit transfer into another province | **Province admin of the province RECEIVING it**, or super-admin / national support |
| Raise any other request | Any authenticated user |

Two asymmetries are deliberate:

- **Filling a vacancy is routine; removing a sitting officer is not.** A
  province admin can appoint into an empty post directly, but displacing someone
  goes through an approval.
- **An officer may appoint any office at their own level, including one that
  outranks them.** A `prov-admin` may appoint the `picp` of their province. This
  was chosen explicitly over restricting administrative officers to
  administrative roles.

Nobody may approve a request they raised themselves. (A super-admin is exempt;
national support is deliberately not — the four-eyes rule holds for them.)

---

## Seeing your officers

### GET /v1/principal-officers/roster

Every office at a unit — **filled and vacant** — with the holder's contact
details. A list of appointments cannot show a post nobody holds, and an empty
post is exactly what an officer needs to act on.

```
GET /v1/principal-officers/roster?levelType=province&scopeCode=LA47
Authorization: Bearer <token>
```

Omit both parameters and you get every unit you have standing at. A super-admin
**must** name a unit — they have no home one.

```json
{
  "totalCount": 7, "filled": 5, "vacant": 2,
  "records": [
    {
      "levelType": "province", "scopeCode": "LA47",
      "roleSlug": "prov-admin", "roleName": "PROVINCE ADMIN",
      "vacant": false,
      "appointmentId": "65a1f0000000000000000aaa",
      "appointedAt": "2026-09-03T09:14:02.113Z",
      "appointmentType": "substantive",
      "holder": {
        "userId": "65a1…", "username": "aadeyemi", "name": "A Adeyemi",
        "email": "a.adeyemi@example.org", "phone": "+2348012345678",
        "whatsApp": "+2348012345678", "avatarUrl": ""
      },
      "canAppoint": true
    },
    {
      "levelType": "province", "scopeCode": "LA47",
      "roleSlug": "picp", "roleName": "PIC PROVINCE",
      "vacant": true,
      "appointmentId": "", "appointedAt": null, "appointmentType": "",
      "holder": null,
      "canAppoint": true
    }
  ]
}
```

> **Contact details are read live from the user record**, not copied onto the
> appointment — a denormalised phone number goes stale the moment it is updated.

`canAppoint` tells the UI whether to offer an "appoint" button per row, so it
never has to guess the authority rules.

`403` when you ask for a unit you have no standing at; the response includes
`yourUnits` so the client can correct itself.

---

## Appointing at your own level

`POST /v1/principal-officers` is **no longer super-admin only**. An officer may
appoint within their own unit.

```
POST /v1/principal-officers
{ "userId": "65a1…", "roleSlug": "prov-accountant" }
```

The unit is derived from the role's `level_type` read off the target user's
profile — omit `scopeCode` and it is their own province. Authorisation is
checked **after** the unit is resolved, because that is the first moment it is
known.

| Status | When |
|---|---|
| `201` | appointed |
| `403` | `NO_STANDING_AT_UNIT` — outside your unit |
| `409` | `OFFICE_ALREADY_HELD` — use the transfer request instead |
| `400` | unknown user/role, role not flagged as an office, user deactivated |

Full field reference in `PRINCIPAL_OFFICERS_DOCS.md`.

---

## The approval workflow

One collection, `approvalRequests`, serves all three request types — the shape
is identical (raise, decide, execute, record) and only the payload and approver
rule differ.

```
PENDING ──approve──▶ APPROVED   (the change has been performed)
   │                 └─ or FAILED if the change itself failed
   ├────reject────▶ REJECTED
   └────cancel────▶ CANCELLED
```

> **A request is not the source of truth.** It records what was *asked for*.
> Approving it performs the change through the same services an admin would use
> directly, and stores the outcome in `executionResult`. If the change fails
> after approval the request is marked **`FAILED`**, not `APPROVED` — an approval
> that silently did nothing is worse than a rejection, because it looks done.

At most **one PENDING request per person per type**, enforced by a partial unique
index. Without it two admins raising the same transfer produces two approvals,
the second executing against a world the first already changed.

---

### POST /v1/approvals/officer-transfer

Replace the holder of an office. **Approved by a super-admin.**

```json
{ "appointmentId": "65a1f0000000000000000aaa", "toUserId": "65a1…", "reason": "Posted to Region 7" }
```

On approval this runs the same ordered end-then-appoint as a direct transfer,
**including the rollback** — if the successor turns out to be ineligible the
incumbent is reinstated.

### POST /v1/approvals/officer-promotion

Move someone into a different office, optionally ending the one they hold.
**Approved by a super-admin.**

```json
{ "toUserId": "65a1…", "roleSlug": "picp", "appointmentId": "65a1…(the post they are leaving)", "reason": "Promotion" }
```

`scopeCode` is optional — omitted, the office is their own unit at that level.
`appointmentId` is optional: omit it for a promotion into a post from no post.

### POST /v1/approvals/user-transfer

Move a user to another parish. **Approved by a province admin of either province
involved, or a super-admin.**

```json
{ "userId": "65a1…", "toParishCode": "211003", "reason": "Relocated" }
```

The destination chain is resolved **at request time**, so an unknown or
`DEPARTMENT` parish is refused when it is raised rather than surprising the
approver later.

On approval:

1. All seven hierarchy fields and `parishRef` are updated to the new parish's chain.
2. **Offices at units the user has left are vacated**, reason `TRANSFERRED` — an
   office belongs to the unit, not the person. An office at a unit they are
   *still* in (a region post when only the parish changed within that region) is
   left alone.
3. **`users.roles` is not touched.** Stripping roles as a side effect of a
   transfer is exactly the silent edit the cleanup pass was kept separate to
   avoid. They surface on the conflicts report at the new unit instead.

```json
// executionResult
{
  "ok": true,
  "movedFrom": { "province": "LA47", "parish": "211549", "…": "…" },
  "movedTo":   { "province": "LA99", "parish": "211003", "…": "…" },
  "officesVacated": [
    { "appointmentId": "65a1…", "roleSlug": "prov-admin", "levelType": "province", "scopeCode": "LA47" }
  ],
  "rolesUnchanged": "[\"prov-admin\",\"pic-parish\"]"
}
```

---

### POST /v1/approvals/unit-transfer

Move a **parish, area or zone into another province**. The one request whose
*raising* is gated — the requester must be a province admin of the province the
unit is **leaving** — and the one whose approver is fixed to the other side:
**a province admin of the province RECEIVING the unit**, or a super-admin /
national support. The giving province cannot approve its own request.

```json
{ "level": "parish", "unitCode": "211343", "toParentCode": "AR8000000999", "reason": "Relocated" }
```

> **The receiving administrator does not need this request.** Since 2026-09-18
> an administrator of the *receiving* province (`prov-admin`, or a `reg-admin` /
> `sub-cont-admin` / `cont-admin` above it) may pull the unit in directly with
> `POST /v1/hierarchy-transfers/transfer` — no request, no second approver. This
> request exists for the **giving** side: the province a unit is leaving cannot
> push it out alone, so it asks and the receiver approves. If both happen — LA10
> raises, and LA133's administrator pulls through `/transfer` before approving
> here — the pull **settles the request**: it ends `APPROVED`, `decidedBy` that
> administrator, `decisionNote` "Accepted by the receiving administrator directly
> through /v1/hierarchy-transfers/transfer", `executionResult.settledBy:
> "transfer"` and `executionResult.placedAt` the area the administrator chose,
> which wins over the `toParentCode` the request named. A pull into a province
> other than the one the request names is refused `409 REQUEST_PENDING_ELSEWHERE`
> until the request is decided or cancelled.

`level` is `parish`, `area` or `zone`. The destination's level is **derived** —
one rank above `level` — and is never sent, exactly as in
`POST /v1/hierarchy-transfers/transfer`. A province or anything above it is moved
by a super-admin through `/v1/hierarchy-transfers/admin/move`.

**How you get here.** A province admin who calls `/transfer` on a move whose
destination is in another province gets `403 TRANSFER_NEEDS_APPROVAL`, and
`detail.requestBody` is exactly the body to POST here (`detail.requestEndpoint`
names this path). `GET /hierarchy-transfers/preview` says the same thing in
`permissionCode` and `approvalHint`, before the user clicks anything.

Both chains are resolved **at request time**, through the same planner
`/transfer` uses, so:

- a same-province move is refused (`400 SAME_PROVINCE`) — use `/transfer`;
- an unknown unit, or one already under the destination, is refused then
  (`EMPTY_UNIT`, `ALREADY_THERE`), not at approval;
- the approver sees what will happen in `planSnapshot` — both chains, the
  inherited fields, and whether either end is split.

On approval the unit is **re-planned**; the snapshot is never trusted. If either
province has changed since the request was raised — someone moved the destination
area into a third province — the request lands `FAILED` with
`executionResult.code = "STALE_REQUEST"` and nothing is written. Otherwise the
move runs through the same code path as `/transfer`: the unit keeps its own
code, inherits every code above it from the destination, every parish and user
beneath it follows, and stranded offices are vacated. The hierarchy job records
`approvalRequestId`, and its `authorisedVia` names the request, the requester
and the approver — never `"super-admin"`.

```json
// executionResult
{
  "ok": true,
  "jobId": "66b2…",
  "authorisedVia": "approval 66b1… — requested by admin.la47 (province LA47), approved by admin.la99 (province LA99)",
  "level": "parish",
  "unit": { "code": "211343", "name": "…" },
  "toParentLevel": "area",
  "toParentCode": "AR8000000999",
  "fromProvince": "LA47",
  "toProvince": "LA99",
  "cascade": { "membersUpdated": 1, "usersUpdated": 14, "officesVacated": [] },
  "sourceWasSplit": false,
  "destinationWasSplit": false,
  "warnings": []
}
```

The receiving province's inbox:

```
GET /v1/approvals?requestType=UNIT_TRANSFER&status=PENDING&toProvince=LA99
```

---

### Deciding

```
POST /v1/approvals/:id/approve    { "decisionNote": "Confirmed with the RPO" }
POST /v1/approvals/:id/reject     { "decisionNote": "Wrong parish code" }
POST /v1/approvals/:id/cancel
```

`approve` **performs the change** and returns the request with
`status`, `decidedBy`, `decidedAt` and `executionResult`.

The request is **claimed atomically** first — `PENDING → EXECUTING` in one
write — so two approvers arriving together cannot both execute it; the second is
told it was decided a moment ago. The same claim guards `reject` and `cancel`,
so neither can land on a request another admin is executing that instant.

`cancel` is for the person who raised the request, or a super-admin / national
support.

| Request status | Meaning |
|---|---|
| `PENDING` | awaiting a decision |
| `EXECUTING` | claimed by an approver and being applied — if it stays here, execution crashed and a human is needed |
| `APPROVED` | applied; see `executionResult` |
| `FAILED` | approved, but the change failed; see `executionResult.error` / `.code` |
| `REJECTED` / `CANCELLED` | nothing changed |

| Status | When |
|---|---|
| `403` | `NOT_AUTHORISED_TO_DECIDE` — wrong approver for this type, or approving your own request |
| `400` | already decided, or the change failed (request left `FAILED`) |
| `409` | `OFFICE_ALREADY_HELD` — someone took the office between raise and approve |

### Listing — the inbox

```
GET /v1/approvals                                  # everything, untreated first
GET /v1/approvals?decidable=true                   # only what I can decide now
GET /v1/approvals?treated=false                    # awaiting an outcome, anyone's
GET /v1/approvals?treated=true&sortBy=decidedAt    # history, most recently decided first
GET /v1/approvals?requestType=USER_TRANSFER&scopeCode=LA47&pageNo=1&pageSize=20
GET /v1/approvals/:id
```

**Default order is `sortBy=pending`**: untreated requests first, newest first
within each group. What needs attention never sinks under what has been decided.
`sortBy` also takes `createdAt`, `requestedAt`, `decidedAt`, `requestType` or
`status`, with `sortDir=asc|desc` (default `desc`). Paging is stable under every
sort.

Filter by `requestType`, `status`, `subjectUserId`, `scopeCode`, `levelType`,
`fromProvince`, `toProvince`, `subjectUnitCode`, `subjectLevel`, and two derived
filters:

| Filter | Meaning |
|---|---|
| `treated=false` | `PENDING` or `EXECUTING` — no outcome yet |
| `treated=true` | `APPROVED`, `REJECTED`, `CANCELLED` or `FAILED` |
| `decidable=true` | only requests **the caller** may decide right now; implies `treated=false` |

**Every row carries three derived fields**, so the UI keeps no status map of its own:

| Field | Meaning |
|---|---|
| `treated` | `false` while the request awaits an outcome |
| `canDecide` | `true` when the **caller** may approve or reject this one now — the same rule `approve` enforces, including the self-approval rule. Show the buttons on this, not on role. |
| `requestTypeLabel` | `Officer transfer`, `Officer promotion`, `User transfer`, `Unit transfer` |

`GET /v1/approvals/:id` returns the same three fields.

The response also carries a `summary` over the current filter, **ignoring any
status filter**, so tabs can show counts whichever tab is open:

```json
{
  "totalCount": 3,
  "sortBy": "pending",
  "sortDir": "desc",
  "pageNo": 1,
  "pageSize": 20,
  "summary": {
    "total": 41, "untreated": 3, "treated": 38, "decidable": 2,
    "byStatus": { "PENDING": 3, "APPROVED": 30, "REJECTED": 6, "FAILED": 2 },
    "byRequestType": { "USER_TRANSFER": 25, "UNIT_TRANSFER": 4, "OFFICER_TRANSFER": 8, "OFFICER_PROMOTION": 4 }
  },
  "records": [
    { "id": "…", "requestType": "UNIT_TRANSFER", "requestTypeLabel": "Unit transfer",
      "status": "PENDING", "treated": false, "canDecide": true, "toProvince": "LA99", "…": "…" }
  ]
}
```

`decidable` is evaluated in code, not as a query — the approver rule depends on
the caller's standing against each request — so that filter fetches the untreated
set and pages it in memory. The set is small by construction: the unique pending
index allows one open request per subject.

---

## Removing a secondary role

**No new endpoint** — this already exists:

```
POST /v1/secondary-roles/:id/end     { "endedReason": "No longer covering" }
```

It deactivates the grant, keeps the row as history, frees the unique index for a
re-grant, and **immediately drops any session acting as it** — without that, the
granted chain sits inside the holder's access token and the revocation would not
bite for up to ten days.

Nothing is hard-deleted anywhere in this module, by design: "who managed this
parish last year" has to stay answerable.

---

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `NO_STANDING_AT_UNIT` | `403` | Appointing outside your own unit |
| `NOT_AUTHORISED_TO_DECIDE` | `403` | Wrong approver, or your own request |
| `REQUEST_ALREADY_PENDING` | `409` | One pending request per person (or unit) per type |
| `OFFICE_ALREADY_HELD` | `409` | The office has a sitting holder |
| `NOT_OFFICE_HOLDER` | `409` | `switch-role` under `enforce` |
| `TRANSFER_NEEDS_APPROVAL` | `403` | From `/hierarchy-transfers/transfer`: you hold the unit's province but the destination is in another. `detail.requestBody` is the body to POST to `/v1/approvals/unit-transfer` |
| `NO_STANDING_OVER_SOURCE` | `403` | Raising a unit transfer without province standing over the province the unit is leaving |
| `SAME_PROVINCE` | `400` | Both ends of a unit transfer are in one province — use `/transfer` |
| `LEVEL_NOT_REQUESTABLE` | `400` | A province or above cannot be requested — `/admin/move` |
| `PROVINCE_UNRESOLVED` | `409` | The unit or destination has no province on its rows |
| `STALE_REQUEST` | — | In `executionResult.code` of a `FAILED` unit transfer: a province changed between raise and approve |

---

## Frontend guidance

1. **Roster is the officer's home screen.** Render `vacant` rows differently —
   they are an action, not an absence. Use `canAppoint` per row rather than
   re-deriving the rules.
2. **Never offer "appoint" over a sitting officer.** The roster tells you the post
   is filled; offer *request transfer* instead, so the user meets the approval
   flow rather than a `409`.
3. **Show `executionResult` after an approval**, especially `officesVacated` — a
   transfer that quietly removed someone's provincial post is something the
   approver should see immediately.
4. **Treat `FAILED` distinctly from `REJECTED`.** Rejected means someone said no.
   Failed means someone said yes and it did not work — that needs a human.
5. **Poll `GET /v1/approvals?decidable=true`** for the approver's queue — it is
   exactly the set this user can act on, with the self-approval rule applied.
   Use `summary.decidable` for the badge count. For a combined view, the default
   `sortBy=pending` puts the untreated rows first, and each row's `treated` and
   `canDecide` tell you how to render it. There are no notifications in this
   module yet.
6. **A request being open to anyone is intentional.** Do not hide the "request"
   buttons behind a role check — the authority is on the decision. The one
   exception is the unit transfer, which only the giving province's admin may
   raise; there, follow the server's lead below.
7. **On `403 TRANSFER_NEEDS_APPROVAL`** from `/hierarchy-transfers/transfer`, or
   `permissionCode === "TRANSFER_NEEDS_APPROVAL"` from `/preview`, offer
   "Request transfer" and POST `detail.requestBody` (or `approvalHint.requestBody`)
   to `/v1/approvals/unit-transfer`. Show `detail.toProvince` — that province's
   admin is who will decide.
8. **Treat `EXECUTING` as "in progress", not as a decision.** A request that
   stays there after a refresh is one whose execution crashed; surface it like
   `FAILED`.
