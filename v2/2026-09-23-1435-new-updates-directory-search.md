# One search across people and parishes, live and deleted

`GET /v1/search` and five routes beneath it. One term finds a person or a parish
by name, hierarchy code, phone or email, case-insensitively, with wildcards.
Deleted rows are searchable too, behind the guard the deleted listings already
use.

## What changed and why

Finding someone or somewhere took knowing which endpoint to ask and which field
to name. `POST /v1/users/search` takes a **column name** and a value from the
caller and filters on them — which means the caller chooses the column, and it
will filter on `token`, `otp` or `password` just as readily as on `email`,
confirming a guess by whether a row comes back. `GET /v1/pastors/search` is
safe but deliberately narrow: exact email, exact username, anchored name
prefixes, one collection. Parishes had `POST /v1/parishDirectory/search` and
two `searchAndFilter` variants, none of which reach a user.

None of them answers the question people actually arrive with, which is "I have
this string — a name, a number, a code, part of an email — find it."

So: a dedicated mount where **the caller chooses a term and the server chooses
the columns**. A field not named in `src/components/Directorysearch/service.ts`
cannot be searched, and no secret column is named there. The returned user
projection lists what to **keep** rather than what to strip, so a sensitive
column added to the schema later cannot leak through it.

Deleted rows became searchable at the same time because the listings that exist
(`GET /v1/users/deleted`, `GET /v1/parishDirectory/deleted`) are date-ordered
pages with no way to ask a question. Finding one account among four hundred
deletions meant paging.

## Behaviour changes on deploy

**None.** Every existing search endpoint is untouched and behaves exactly as it
did. This is six new routes on a new mount.

## The routes

| Method | Path | Guard |
|---|---|---|
| GET | `/v1/search` | any signed-in caller |
| GET | `/v1/search/users` | any signed-in caller |
| GET | `/v1/search/parishes` | any signed-in caller |
| GET | `/v1/search/deleted` | super-admin or an elevated role |
| GET | `/v1/search/deleted/users` | super-admin or an elevated role |
| GET | `/v1/search/deleted/parishes` | super-admin or an elevated role |

The live routes take `attachDbUser`, matching `/v1/pastors/search` and the
directory listings: any signed-in caller may look a colleague or a parish up,
and withholding it here would only send them back to those. The deleted routes
take `requireDbElevated`, matching the deleted listings they complement — a
removed account still carries a name, a number and an address, and who may read
that was already decided.

Live and deleted are **separate scopes, never mixed**. A live search that
quietly included deleted parishes would put closed congregations in front of
someone looking for an open one; a deleted search that included live rows would
make a recovery tool lie about what was lost.

## Parameters

| Parameter | Applies to | Matched against |
|---|---|---|
| `q` | both | every field group below, at once |
| `name` | both | user name columns; parish and ancestor names |
| `parishName` | parishes | same as `name`; the two are one parameter under two names |
| `username` | users | `username` |
| `email` | both | user `email`; parish, pastor and assistant-pastor email |
| `phone` | both | user `phone` and `username`; parish, pastor and assistant-pastor phone |
| `code` | both | every hierarchy code |
| `pageNo`, `pageSize` | both | 1-based page, default 20, **capped at 200** |

`q` is OR-ed across everything. The narrowing parameters are AND-ed with each
other and with `q`, so adding one finds fewer rows, never more. Within `name`,
each **word** must match something, so `gr ok` finds Grace Okonkwo and narrows
rather than widening.

Anything else in the query string is ignored — there is no parameter that names
a column.

## How each field is matched

Never case-sensitively, in any shape.

| Field group | Shape | Why |
|---|---|---|
| hierarchy codes | **prefix** | `LA47` is one province; `LA4` is an exploration that should stop at the LA4* provinces |
| names | **contains** | searching a directory for `MERCY` has to find `RCCG CHAPEL OF MERCY` |
| email, username | **contains** | so `@rccg` and a full address both work |
| phone | **variant set, exact** | `0803…`, `234803…` and `+234803…` are one number — the same variant set the migrated lookup uses. Matched against `username` too, because a great many accounts sign in with their number |

### Wildcards

A term carrying `*` or `?` is matched as an **anchored pattern** instead, against
every field in the group:

| Term | Means |
|---|---|
| `ADE*` | starts with ADE |
| `*MERCY*` | contains MERCY |
| `LA4?` | LA4 plus exactly one character — LA47, not LA470 |
| `??` | any two-character value |

Anchored because that is what a wildcard means everywhere else a person has met
one. A wildcard overrides the field's usual shape: the caller has said what they
mean, and silently re-anchoring their pattern would produce a result they cannot
account for.

Everything that is not a wildcard is escaped, so `ST. JOHN (1)` is matched as a
name. `.*` is **not** a match-everything pattern here — the dot is a literal and
only the `*` is a wildcard, so it means "starts with a dot".

### Minimum length

A term under two characters is refused with `400 TERM_TOO_SHORT`, naming the
parameter. A one-character `contains` matches most of the collection, spends the
query budget and answers nothing. A wildcard is exempt: `a*` and `??` are
statements of intent however short.

## Requests and responses

```
GET /v1/search/users?q=08035707056
GET /v1/search/users?name=grace%20okonkwo&code=LA47
GET /v1/search/parishes?q=MERCY&pageSize=50
GET /v1/search/parishes?code=LA4&parishName=chapel
GET /v1/search?q=ADE*
GET /v1/search/deleted/users?email=@gmail
```

A single-collection route returns:

```json
{
  "scope": "live",
  "totalCount": 3,
  "pageNo": 0,
  "pageSize": 20,
  "records": [
    {
      "id": "6520a1…",
      "name": "Grace Okonkwo",
      "firstName": "Grace", "lastName": "Okonkwo", "otherNames": "",
      "username": "grace.o", "email": "grace@example.test",
      "phone": "08035707056", "phoneCode": "234",
      "gender": "F", "designation": "", "status": "1", "userStatus": "ACTIVE",
      "roles": ["pic-parish"],
      "hierarchy": {
        "parish": "211003", "area": "A031", "zone": "Z012",
        "province": "LA47", "region": "R07",
        "subContinent": "ECS0001", "continent": "CNT02"
      },
      "deleted": { "isDeleted": false, "at": "", "by": "", "byUsername": "" }
    }
  ]
}
```

A parish record carries codes **and** names at every level, because a code alone
tells a reader nothing:

```json
{
  "id": "652f…",
  "parishCode": "211003", "parishName": "RCCG CHAPEL OF MERCY",
  "parishAlias": "", "parishType": "", "status": "1",
  "hierarchy": {
    "area": { "code": "A031", "name": "AREA 31" },
    "zone": { "code": "Z012", "name": "ZONE 12" },
    "province": { "code": "LA47", "name": "LAGOS PROVINCE 47" },
    "region": { "code": "R07", "name": "REGION 7" },
    "subContinent": { "code": "ECS0001", "name": "…" },
    "continent": { "code": "CNT02", "name": "…" }
  },
  "contact": { "phone": "08…", "email": "…", "address": "…",
               "city": "…", "state": "…", "country": "NG" },
  "pastor": { "name": "…", "phone": "…", "email": "…" },
  "deleted": { "isDeleted": false, "at": "", "by": "",
               "reason": "", "reactivateAt": "" }
}
```

The `deleted` block is **present and empty on a live row** rather than absent, so
one shape serves both scopes and a client needs no branch.

`GET /v1/search` and `GET /v1/search/deleted` return both collections, each paged
and counted separately:

```json
{
  "scope": "live", "pageNo": 0, "pageSize": 20,
  "users":    { "totalCount": 3,  "records": [ … ] },
  "parishes": { "totalCount": 12, "records": [ … ] }
}
```

Two answers to one question, not one merged list. The collections share no field
that could order them against each other, and callers show them in separate
sections anyway.

## Errors

| Status | Code | When |
|---|---|---|
| 400 | `NOTHING_TO_SEARCH` | no `q` and no narrowing parameter |
| 400 | `TERM_TOO_SHORT` | a non-wildcard term under two characters; the message names the parameter |
| 403 | — | a deleted route without super-admin or an elevated role |

## What is logged

The activity line records **which parameters** were used and how many rows came
back. Never the term, never the results. A search for a person is itself a fact
about that person, and copying the term into the log would put names, numbers
and addresses into a second collection nobody thinks of as holding them.

Activities: `SEARCH_DIRECTORY` for live, `SEARCH_DELETED` for deleted.

## Configuration

None. No new environment variables, no new indexes, no deployment step.

## Known gaps

**This scans; it does not seek.** A case-insensitive regex cannot take tight
index bounds — the index is stored case-sensitively, so `/^la4/i` scans the whole
index exactly as `/mercy/i` does. Anchoring buys fewer *matched* documents, not a
cheaper scan. Over ~53,000 parishes and ~57,000 users that is still worth having,
and the page cap of 200 plus the two-character minimum keep the worst case
bounded, but nothing here is a seek.

Making it one needs a normalised lowercase column per searched field with its own
index, or a collation index. Neither is built. If these routes become hot, that
is the work — not more parameters.

**Results are not scoped to the caller's unit.** Any signed-in caller can find
any live person or parish, which is what `/v1/pastors/search` and the directory
listings already allow. If the organisation wants directory search narrowed to
standing, that is a deliberate product decision and a separate change; it would
also make these routes considerably cheaper.

**`POST /v1/users/search` is still open.** It still takes a column name from the
caller, still has no role guard, and still accepts `token`, `otp` and `password`
as search columns. This change does not touch it. It should be given a column
whitelist or retired in favour of these routes.
