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

`super-admin` is unbounded and is the only exception.

| Action | Who |
|---|---|
| See the officers of a unit | Anyone with standing at that unit; super-admin anywhere |
| Appoint to a **vacant** office | Anyone with standing at that unit; super-admin anywhere |
| End or transfer a **sitting** officer | Super-admin — or a request approved by one |
| Approve an officer transfer / promotion | **Super-admin only** |
| Approve a user parish transfer | **Province admin** of either province involved, or super-admin |
| Raise any request | Any authenticated user |

Two asymmetries are deliberate:

- **Filling a vacancy is routine; removing a sitting officer is not.** A
  province admin can appoint into an empty post directly, but displacing someone
  goes through an approval.
- **An officer may appoint any office at their own level, including one that
  outranks them.** A `prov-admin` may appoint the `picp` of their province. This
  was chosen explicitly over restricting administrative officers to
  administrative roles.

Nobody may approve a request they raised themselves.

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

### Deciding

```
POST /v1/approvals/:id/approve    { "decisionNote": "Confirmed with the RPO" }
POST /v1/approvals/:id/reject     { "decisionNote": "Wrong parish code" }
POST /v1/approvals/:id/cancel
```

`approve` **performs the change** and returns the request with
`status`, `decidedBy`, `decidedAt` and `executionResult`.

`cancel` is for the person who raised the request, or a super-admin.

| Status | When |
|---|---|
| `403` | `NOT_AUTHORISED_TO_DECIDE` — wrong approver for this type, or approving your own request |
| `400` | already decided, or the change failed (request left `FAILED`) |
| `409` | `OFFICE_ALREADY_HELD` — someone took the office between raise and approve |

### Listing

```
GET /v1/approvals?status=PENDING&requestType=USER_TRANSFER&scopeCode=LA47&pageNo=1&pageSize=20
GET /v1/approvals/:id
```

Newest first. Filter by `requestType`, `status`, `subjectUserId`, `scopeCode`,
`levelType`.

---

## Removing a secondary role


```
POST /v1/secondary-roles/:id/end     { "endedReason": "No longer covering" }
```


---

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `NO_STANDING_AT_UNIT` | `403` | Appointing outside your own unit |
| `NOT_AUTHORISED_TO_DECIDE` | `403` | Wrong approver, or your own request |
| `REQUEST_ALREADY_PENDING` | `409` | One pending request per person per type |
| `OFFICE_ALREADY_HELD` | `409` | The office has a sitting holder |
| `NOT_OFFICE_HOLDER` | `409` | `switch-role` under `enforce` |

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
5. **Poll `GET /v1/approvals?status=PENDING`** for the approver's queue. There are
   no notifications in this module yet.
6. **A request being open to anyone is intentional.** Do not hide the "request"
   buttons behind a role check — the authority is on the decision.
