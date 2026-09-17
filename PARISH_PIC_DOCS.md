# Pastor in Charge — endpoint guide

Who leads a parish, how they are appointed, and why holding the pastor role is
not the same thing.

## Read this first

**Holding `pic-parish` does not make someone the pastor in charge of a parish.**
51,551 accounts hold that role. It names no parish — it is read against whatever
sits in `users.parish` — so a pastor who was transferred, or replaced, or granted
it years ago and never appointed, all still hold it.

`parishPicHolders` is the record of who actually leads where. A parish with no
active row in it has **no** pastor in charge, and that is a real answer rather
than a gap in the data.

| Group | Endpoints | Who calls them |
|---|---|---|
| **Reading** | who leads this parish, list appointments, parish search with pastors | Any signed-in user |
| **Appointing** | appoint, end an appointment | Administrators over a unit containing the parish |

---

## Contents

- [Who can appoint](#who-can-appoint)
- [Appoint a pastor in charge](#appoint-a-pastor-in-charge)
- [End an appointment](#end-an-appointment)
- [Who leads this parish](#who-leads-this-parish)
- [List appointments](#list-appointments)
- [Parish search with pastor data](#parish-search-with-pastor-data)
- [When a pastor leaves](#when-a-pastor-leaves)
- [Only the current pastor may act](#only-the-current-pastor-may-act)
- [Principal officers — the role follows too](#principal-officers--the-role-follows-the-appointment-too)
- [Who decides an officer appointment](#who-decides-an-officer-appointment)
- [Change history](#change-history--the-four-records-read-together)
- [Finding a person](#finding-a-person)
- [All error codes](#all-error-codes)
- [Frontend guidance](#frontend-guidance)
- [What this does not do](#what-this-does-not-do)

---

## Who can appoint

Can the person on the left appoint the pastor in charge of a parish?

| Actor | A parish in their unit | A parish outside it |
|---|---|---|
| Super Admin | ✅ | ✅ |
| National Support | ✅ | ✅ |
| Sub-Continent Admin | ✅ | ❌ |
| Regional Admin | ✅ | ❌ |
| Province Admin | ✅ | ❌ |
| Zone Admin | ✅ | ❌ |
| Area Admin | ✅ | ❌ |
| Parish Pastor (`pic-parish`) | ❌ | ❌ |

**A parish cannot appoint its own head.** Parish level is deliberately absent —
every level that may appoint sits above the parish.

**Containment is judged on the parish's real ancestry**, read from the directory,
never from codes in the request. An area admin passes on the area, a province
admin on the province. Nobody can assert a province they are not in.

---

## Appoint a pastor in charge

```
POST /v1/parish-pics
```

**Who** An administrator of a unit containing the parish, super admin, or
national support. Bearer token.

### Request

```json
{
  "parishCode": "PA015520",
  "userId": "64b7f0c2f1a2b3c4d5e6f701",
  "note": "Appointed at the provincial council, September 2026"
}
```

| Field | Required | Rule |
|---|---|---|
| `parishCode` | **yes** | The parish they will lead |
| `userId` | **yes** | The pastor. Must already belong to this parish |
| `note` | no | Free text, up to 500 characters, kept on the record |

> **The pastor must already have been transferred to the parish.** Their own
> record must name it. The parish code in this request is the parish being filled,
> not a claim about where the pastor is.

### Response — 201

```json
{
  "appointmentId": "64c1a0d3e2b4c5d6e7f80912",
  "authorisedVia": "province PR0042",
  "parishCode": "PA015520",
  "parishName": "RCCG HOUSE OF PRAYER",
  "pastor": {
    "userId": "64b7f0c2f1a2b3c4d5e6f701",
    "username": "grace.okonkwo",
    "name": "Grace Okonkwo"
  },
  "roleAdded": true,
  "roleSlug": "pic-parish",
  "staleChainFields": [],
  "warnings": []
}
```

**`roleAdded` tells you whether the role had to be given.** `true` means they did
not hold `pic-parish` and now do. `false` means they already held it and nothing
was written. Either way the appointment stands — **this is one operation, not an
appointment followed by a trip to user administration.**

`authorisedVia` names the unit the permission came through, and is the same
string written to the audit log. Super admin gets `super-admin`, national support
`elevated:nat-support`.

### `staleChainFields`

```json
{
  "staleChainFields": ["province", "region"],
  "warnings": [
    "the pastor's own record disagrees with the parish on province, region — their copy of the hierarchy is stale, which a transfer would correct"
  ]
}
```

The appointment **succeeded**. Their `parish` matched, which is what the rule
requires, but their denormalised copy of the levels above it disagrees with the
parish row. That is a data fault worth seeing — it usually means they were moved
by something that did not update the whole chain. Show the warning; do not block.

### Response — 403

```json
{
  "status": 403,
  "code": "PIC_NOT_PERMITTED",
  "message": "Parish PA015520 is not inside a unit you administer, so appointing its pastor in charge is not yours to do. It sits in area AR0301, zone ZN0140, province PR0042. You hold: province PR0099.",
  "detail": {
    "parishChain": { "area": "AR0301", "zone": "ZN0140", "province": "PR0042", "region": "R11" }
  }
}
```

### Response — 400 and 409

```json
{
  "status": 400,
  "code": "PASTOR_NOT_IN_PARISH",
  "message": "grace.okonkwo has not been transferred to parish PA015520 — their record still puts them in parish PA009911. Move them first; a pastor cannot be put in charge of a parish they do not belong to.",
  "detail": {
    "pastorParish": "PA009911",
    "targetParish": "PA015520",
    "useInstead": [
      { "useEndpoint": "/v1/approvals/user-transfer", "useBody": null,
        "why": "transfer the pastor to this parish first" }
    ]
  }
}
```

```json
{
  "status": 409,
  "code": "PIC_SEAT_TAKEN",
  "message": "Parish PA015520 already has a pastor in charge — Daniel Eze. End that appointment before making another.",
  "detail": { "holder": { "userId": "…", "username": "daniel.eze", "appointmentId": "…" } }
}
```

### Use case

> A province admin appoints a pastor who was transferred to the parish last week.
> One call: the appointment is recorded, the `pic-parish` role is added, and the
> audit log says who did it and through which unit. Nobody has to go and edit the
> user's roles afterwards.

---

## End an appointment

```
POST /v1/parish-pics/{id}/end
```

**Who** The same people who may appoint there. Authority is judged on the parish
the appointment is for, re-read from the directory — an appointment id does not
imply permission to end it.

### Request

```json
{ "reason": "RESIGNED" }
```

`reason` is one of `PROMOTED`, `TRANSFERRED`, `DEMOTED`, `REMOVED`, `RESIGNED`,
or omitted.

### Response — 200

```json
{
  "appointmentId": "64c1a0d3e2b4c5d6e7f80912",
  "parishCode": "PA015520",
  "endedReason": "RESIGNED",
  "vacant": true
}
```

**No successor is appointed.** Who leads a parish is a decision for someone with
standing, not a consequence of someone else leaving. The parish stays vacant
until somebody appoints.

**Nothing is deleted.** The row is closed with an end date, so succession stays
answerable and the seat is freed for the next holder.

---

## Who leads this parish

```
GET /v1/parish-pics/parish/{parishCode}
```

**Who** Any signed-in user.

### Response — 200, held

```json
{
  "parishCode": "PA015520",
  "pic": {
    "userId": "64b7f0c2f1a2b3c4d5e6f701",
    "username": "grace.okonkwo",
    "name": "Grace Okonkwo",
    "email": "grace.okonkwo@rccg.org",
    "phone": "08031234567",
    "appointmentId": "64c1a0d3e2b4c5d6e7f80912",
    "since": "2026-09-17T09:14:02.000Z"
  }
}
```

### Response — 200, vacant

```json
{ "parishCode": "PA015520", "pic": null }
```

**`null` is the answer, not an error.** A parish with no pastor in charge is a
normal state, and 200 is the honest status for a question that was answered.

---

## List appointments

```
GET /v1/parish-pics?parishCode=PA015520
GET /v1/parish-pics?userId=64b7f0c2f1a2b3c4d5e6f701
GET /v1/parish-pics?active=false&pageNo=1&pageSize=20
```

**Who** Any signed-in user.

Current appointments only unless `active=false`, which returns the ended ones —
the tenure history of a parish, or of a person.

### Response — 200

```json
{ "totalCount": 3, "records": [ … ], "pageNo": 0, "pageSize": 20 }
```

The standard envelope. `pageNo` echoes the internal zero-based value.

---

## Parish search with pastor data

```
POST /v1/parishdirectory/searchAndFilterWithPastorData
```

**Who** The same as the existing parish search.

Everything `searchAndFilter` does, plus a `pic` block on each parish.

### Request

The existing search body, unchanged:

```json
{ "orAnd": "and", "params": [ { "columnName": "provinceCode", "columnValue": "PR0042" } ] }
```

Optional query parameters: `hasPic=true`, `hasPic=false`, `requestedFields`,
`pageNo`, `pageSize`.

### Response — 201

```json
{
  "totalCount": 240,
  "records": [
    {
      "parishCode": "PA015520",
      "parishName": "RCCG HOUSE OF PRAYER",
      "provinceCode": "PR0042",
      "pic": {
        "userId": "…", "username": "grace.okonkwo", "name": "Grace Okonkwo",
        "email": "grace.okonkwo@rccg.org", "phone": "08031234567"
      }
    },
    { "parishCode": "PA015521", "parishName": "RCCG CHAPEL OF LIGHT", "pic": null }
  ],
  "pageNo": 0,
  "pageSize": 20,
  "picFilterApplied": "",
  "picFilterNote": ""
}
```

> **`hasPic` filters the returned page, not the query.** The appointment lives in
> another collection, so a filter inside the parish query cannot see it.
> `totalCount` counts parishes matching the **search**, before the pastor filter,
> and a filtered page can come back short. The response says so in
> `picFilterNote` when the filter is in use.

The pastor comes from the appointment register, **not** from the
`parishPastorName` / `parishPastorPhone` / `parishPastorEmail` strings on the
parish row. Those are free text written by hand at creation and read only by the
birthday cron; nothing keeps them in step with reality.

This is a **new** endpoint. `searchAndFilter` is unchanged and its callers are
untouched.

---

## When a pastor leaves

**Moving a pastor out of a parish ends their appointment. Nobody is appointed in
their place.**

Two paths move a person, and both do this:

- `POST /v1/approvals/user-transfer`, once approved
- `PATCH /v1/users/{id}` with a new `parish`

Both write a `PIC_VACATED` line to the activity log naming the parish and who
caused it.

> **Moving the parish itself does not end anything.** A parish that moves between
> areas keeps its own code and its own pastor — same parish, same leader. Only the
> *pastor* moving vacates the seat.

---

## Only the current pastor may act

`PIC_ENFORCEMENT` decides whether the register is consulted when somebody acts as
a parish's pastor.

| Value | Behaviour |
|---|---|
| `off` (default) | Nothing is looked up. No cost, no change. |
| `warn` | The register is read and a refusal is logged, but the action proceeds. |
| `enforce` | Someone who is not the current holder is refused. |

**Ship in `warn` and read the log before flipping.** With 51,551 holders of a
role that names no parish, `enforce` on day one refuses real people. The point of
the flag is that you find out who first.

An unreadable or misspelt value means `off`. The dangerous direction here is
accidentally enforcing.

---

## All error codes

| Code | Status | Endpoint | Meaning |
|---|---|---|---|
| `PIC_NOT_PERMITTED` | 403 | appoint, end | The parish is not in a unit you administer |
| `PASTOR_NOT_IN_PARISH` | 400 | appoint | Transfer them to the parish first |
| `PARISH_NOT_FOUND` | 400 | appoint | No parish with that code |
| `PARISH_REQUIRED` | 400 | appoint | No parish code sent |
| `USER_REQUIRED` | 400 | appoint | No userId sent |
| `USER_NOT_FOUND` | 400 | appoint | No user with that id |
| `PASTOR_NOT_ACTIVE` | 400 | appoint | The account is inactive, suspended or deleted |
| `PIC_SEAT_TAKEN` | 409 | appoint | The parish already has a pastor in charge |
| `ALREADY_PIC` | 409 | appoint | That person already leads this parish |
| `APPOINTMENT_NOT_FOUND` | 400 | end | No active appointment with that id |
| `INVALID_REQUEST` | 400 | appoint | The body failed validation |

---

## Frontend guidance

**Treat `pic: null` as a normal state.** Show "No pastor in charge" and, for
someone who may appoint, the button. It is not an error and not missing data.

**`roleAdded` is worth showing once.** "Grace Okonkwo is now pastor in charge of
RCCG House of Prayer, and has been given the parish pastor role" tells an
administrator the second step they used to do by hand has happened.

**Read `staleChainFields`.** The appointment worked, but the pastor's record
disagrees with the parish above parish level. Surface it as a data warning on the
person, not as a failure of the appointment.

**`PASTOR_NOT_IN_PARISH` has a next step in `detail.useInstead`.** Offer
"Transfer this pastor first" rather than showing the refusal and stopping.

**Do not use `hasPic` for counting.** It narrows the page after the search has
run. For "how many parishes have no pastor", ask for the page and read
`picFilterNote` — or count from `/v1/parish-pics` instead.

**`parishPastorName` on the parish row is not the pastor in charge.** It is
legacy free text. Read `pic`.

---

## Principal officers — the role follows the appointment too

The same one-operation rule now applies to principal offices, which had the same
two-step problem: appoint through `POST /v1/principal-officers`, then go to user
administration and add the role by hand.

Appointing now writes the entitlement as well, and the appointment records
whether it did:

```json
{
  "roleSlug": "prov-admin",
  "levelType": "province",
  "scopeCode": "PR0042",
  "source": "primary",
  "roleSynced": true
}
```

**`roleSynced` is false for every SECONDARY appointment, and that is correct.**

| `source` | What the entitlement is | `users.roles` |
|---|---|---|
| `primary` | The role on their own record, read against their own unit | the slug is added |
| `secondary` | A grant tying the role to another unit | **untouched** |

`users.roles` is read against the holder's **own** profile geography. Writing
`prov-admin` there for someone appointed over a province they do not belong to
would hand them prov-admin over their own province as well — a second, unasked
authority. The grant already is the entitlement for a secondary appointment, so
there is nothing to add.

`roleSynced: false` on a **primary** appointment means one of two things: they
already held the role, or the role write did not happen. The appointment row is
written first and the role follows, so a crash between them leaves exactly that —
findable, and fixed by appointing again.

---

## Who decides an officer appointment

Raising a principal-officer request and deciding it are different powers. Anyone
with standing may raise one; deciding it happens **from above the office**.

| Approver | A province office in their region | A province office elsewhere | A region office |
|---|---|---|---|
| Super Admin | ✅ | ✅ | ✅ |
| National Support | ✅ | ✅ | ✅ |
| Region Admin (R07) | ✅ | ❌ | ❌ |
| Province Admin (LA47) | ❌ *even their own* | ❌ | ❌ |
| Area Admin | ❌ | ❌ | ❌ |

**Two conditions, both required.** The approver must stand at a level *strictly
senior* to the office, **and** their unit must contain it. Seniority alone would
let a regional admin decide an appointment in another region; containment alone
would let a province admin decide a province office — which is precisely the
conflict of interest the approval step exists to prevent.

This is why **a province admin cannot decide a province-level appointment even in
their own province**. It is not an oversight in the scope check; it is the point.

### It is judged on a snapshot

The office's ancestry is resolved and stored on the request when it is **raised**,
beside `planSnapshot`. Deciding therefore asks about the hierarchy as it stood
when the request was made, not as it stands days later when somebody gets to it.

**A request raised before this existed has no snapshot and stays an unbounded
decision.** That is deliberate: it was raised under the old rule, and opening it
up retroactively would change the terms after the fact.

### Refusal

```json
{
  "status": 403,
  "code": "NOT_AUTHORISED_TO_DECIDE",
  "message": "An officer change is approved from above the office — an administrator of a level senior to it, whose unit contains it — or by a super-admin or national support. An administrator at the office's own level may not."
}
```

Self-approval is still refused for everyone but a super-admin, senior standing or
not.

---

## Change history — the four records, read together

```
GET /v1/change-history
```

**Who** Any signed-in user. A caller bounded to a unit sees only their own; an
unbounded one sees everything.

History already existed in **four** collections with four shapes, four mounts and
four guards. Answering "why is this parish in that province, and who agreed to
it" meant knowing all four existed and reading them separately. This reads them
and normalises them. **It writes nothing and owns nothing** — a fifth store would
be one more thing to keep in step with the four that are already authoritative.

| `source` | Comes from | Answers |
|---|---|---|
| `approval` | `approvalRequests` | who asked, who decided, why, and what was decided |
| `hierarchy` | `hierarchyChangeJobs` | what a move, promotion, realign or rollback rewrote |
| `principal-office` | `principalOfficeHolders` | who held which office, and when it ended |
| `parish-pic` | `parishPicHolders` | who led which parish, and when it changed |

### Filters

`source` (comma separated), `type`, `status`, `userId`, `unitCode`, `limit`
(max 200).

### Response — 200

```json
{
  "records": [
    {
      "source": "approval",
      "referenceId": "64c1a0d3e2b4c5d6e7f80912",
      "type": "USER_TRANSFER",
      "status": "APPROVED",
      "at": "2026-09-17T09:14:02.000Z",
      "subject": { "userId": "…", "username": "moved.person", "name": "Moved Person" },
      "unit": { "level": "parish", "code": "PA015520", "name": "…" },
      "from": { "province": "PR0042", "parish": "PA009911" },
      "to": { "province": "PR0099", "parish": "PA015520" },
      "requestedBy": { "userId": "…", "username": "asker", "at": "…" },
      "decidedBy": { "userId": "…", "username": "decider", "at": "…", "note": "Agreed" },
      "reason": "Family relocation",
      "detail": { "roleSlug": "", "levelType": "", "scopeCodes": ["PR0042", "PR0099"] }
    }
  ],
  "sources": ["approval", "hierarchy", "principal-office", "parish-pic"],
  "scope": "province PR0042",
  "omittedOutsideYourUnit": 14
}
```

A `hierarchy` entry adds `detail.rowsChanged`, `detail.reversible` and
`detail.rolledBackBy`, so "what did that move actually rewrite, and can it still
be undone" is answerable from the list.

A hierarchy job has no approver of its own. When it was executed from an
approval, `decidedBy.viaApprovalRequest` names it, so the two records join.

### Scope fails closed

> A record reaches a bounded caller **only when it can be positively matched to a
> unit they administer**. Anything whose unit cannot be determined is omitted
> rather than shown.

Across four shapes the ways to be wrong outnumber the ways to be right, and being
wrong means showing somebody another province's business. **A caller with no
administrative unit sees nothing** — that is the correct answer, not an error.

`omittedOutsideYourUnit` says how many were withheld, because a short list
otherwise reads as "nothing happened".

---

## Finding a person

```
GET /v1/pastors/search?phone=08031234567
GET /v1/pastors/search?name=gr%20ok
GET /v1/pastors/search?email=grace.okonkwo@rccg.org
GET /v1/pastors/search?username=grace.okonkwo
GET /v1/pastors/search?parishCode=PA015520
```

**Who** Any signed-in user. Combine as many as you like; they narrow together.

Each identifier is matched the way that identifier is actually stored, rather
than by one generic regex over everything.

| Field | Matching |
|---|---|
| `phone` | The same variant set the migrated lookup uses, matched **exactly** against both `phone` and `username` |
| `email` | Exact, case-insensitive |
| `username` | Exact, case-insensitive — **not** a prefix |
| `name` | Anchored prefix, per word, across first, last and other names |
| `parishCode` | Exact |

**Phone finds the same person from any form.** `08031234567`,
`+2348031234567`, `234 803 123 4567` and `(0803) 123-4567` all resolve to the
same set. It searches `username` as well as `phone` because a great many
accounts log in with their number.

**Name is a prefix, per word, in any order.** `gr ok` finds Grace Okonkwo.
`okonkwo grace` finds her too. **`deyemi` does not find Adeyemi** — the rule is
"starts with", and it is worth telling users that plainly rather than leaving
them to guess why a search came back empty.

### Response — 200

```json
{
  "totalCount": 1,
  "records": [
    {
      "user": {
        "id": "64b7f0c2f1a2b3c4d5e6f701",
        "name": "Grace Okonkwo", "firstName": "Grace", "lastName": "Okonkwo",
        "username": "grace.okonkwo", "email": "grace.okonkwo@rccg.org",
        "phone": "08031234567", "designation": "Pastor",
        "status": "1", "userStatus": "ACTIVE"
      },
      "hierarchy": { "parish": "PA015520", "province": "PR0042", "region": "R11" },
      "roles": ["pic-parish"],
      "pic": { "isPastorInCharge": true, "parishCode": "PA015520", "since": "…" }
    }
  ],
  "pageNo": 0, "pageSize": 20
}
```

**The `pic` block is the point.** It separates holding the role from sitting in
the seat:

```json
"pic": {
  "isPastorInCharge": false,
  "holdsPicRole": true,
  "theirParishIsLedBy": { "userId": "…", "username": "daniel.eze" }
}
```

That reads: they hold `pic-parish`, they do **not** lead a parish, and the parish
they belong to is led by someone else. With 51,551 holders of that role, this is
usually the answer an administrator needs.

### One person, in full

```
GET /v1/pastors/{userId}
```

Returns `user`, `hierarchy`, `roles`, `profile`, `pastorInChargeOf` and
`principalOffices` as **separate blocks**. Separate on purpose: the same fact can
disagree between `users` and `userProfiles`, and merging them would pick a winner
silently.

`migrated.included` is always `false` — the fifteen `jos_*` tables live behind
`/v1/utility/migrated-profile/v3/{phone}` and pulling them into every profile
read would make this the slowest call in the API.

---

## What this does not do

**Nothing is enforced yet.** `PIC_ENFORCEMENT` defaults to `off`, so holding
`pic-parish` still grants whatever it granted before. The register records the
truth; switching to `warn` then `enforce` is a separate, deliberate act.

**No stale roles have been cleaned.** Users holding `pic-parish` without an
appointment keep it. Removing it from thousands of accounts is a migration with
its own review, not a side effect of this work.

**There is no approval workflow.** An administrator with standing appoints
directly. That was a decision: the appointer already holds the unit, and adding a
second signature to an in-unit appointment buys nothing.

**Area and zone administrators can appoint.** They cannot reset passwords, but
they can name a parish's pastor. Those are different powers with different rules.

**`hasPic` cannot filter across pages.** See above. If filtered totals matter,
that needs an aggregation across two collections and is not built.

**One pastor, one parish.** The unique index is on the parish, not the person, so
nothing stops one pastor leading two parishes. If that should be refused, it is a
second index, not a code change.

**A principal-office role can still be assigned by hand while the seat is
vacant.** `vetRoles` refuses a role whose office somebody *else* holds, which is
the case that matters most, but it does not require an appointment to exist
before the role may be given. Closing that would refuse ordinary onboarding, so
it is a decision rather than an oversight.

**Ending a principal office does not remove the role.** The appointment closes;
the entitlement stays. Same reasoning as the parish tier — removing roles from
people is a migration with its own review.

**Change history does not read the activity log.** `activityLogs` is the
catch-all — every login, search and view — and folding it in would bury the four
records that answer "who agreed to this" under traffic. Read it directly at
`/v1/activityLogs` when you want that.

**Change history pages by `limit`, not by page number.** It merges four sorted
lists, so a stable offset across them would mean reading all four in full. Ask
for what you need and filter.

**An officer approval still cannot be decided at the office's own level.** See
[Who decides an officer appointment](#who-decides-an-officer-appointment) — that
is the conflict of interest the rule exists to prevent, and delegating upward did
not change it.

**Name search is a prefix, and it is not a seek.** A case-insensitive regex takes
no tight index bounds even when anchored — measured, both `/^ade/i` and `/ade/i`
plan as an index scan over the whole index. The index keeps the scan off the
documents, which over ~55,000 rows is worth having, but a true seek needs a
normalised lowercase name column or a collation index. Neither is built.

**No contains-anywhere name search.** Adding one is a line of code and would
scan the collection on an endpoint any signed-in user can call. If it is needed,
it should come with the normalised column above, not on its own.

**`users` gained four indexes** — `phone`, `lastName`, `firstName`, `parish` —
because the collection had none beyond the declared uniques. They build on boot
where `autoIndex` is on.

> **Verify the unique indexes on `users` in production.** On a local database
> they are **absent**: `username` and `email` are declared with the
> `mongoose-beautiful-unique-validation` string form (`unique: "message"`), and
> creating an index from that fails — *"not convertible to bool"*. Mongoose
> reports a failed build on an event nobody listens to, so it fails silently.
> Locally that leaves `users` with no unique constraint on either field, enforced
> only by the plugin on `save()` — which `updateOne` and raw collection writes
> bypass. Production may differ if those indexes were ever created by hand.
> `db.users.getIndexes()` settles it in one line.
